create procedure update_biochemistryexpert(IN achievementcategoryid bigint, IN userids bigint[])
    language plpgsql
as
$$
BEGIN
    CALL update_subject_expert('Biochemistry', achievementcategoryid, userids); -- Call the main procedure with 'Biochemistry'
END;
$$;

alter procedure update_biochemistryexpert(bigint, bigint[]) owner to learner;
