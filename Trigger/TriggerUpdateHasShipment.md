create function "TriggerUpdateHasShipment"() returns trigger
    language plpgsql
as
$$
BEGIN
  NEW."hasShipment" := public."DetermineHasShipment"(NEW."id");
    
  RETURN NEW;
END;
$$;

alter function "TriggerUpdateHasShipment"() owner to neetprep_rw;
