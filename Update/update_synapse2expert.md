create procedure update_synapse2expert(IN achievementcategoryid bigint, IN userids bigint[])
    language plpgsql
as
$$
BEGIN
    CALL update_subject_expert('Synapse II', achievementCategoryId, userIds);
END;
$$;

alter procedure update_synapse2expert(bigint, bigint[]) owner to learner;
