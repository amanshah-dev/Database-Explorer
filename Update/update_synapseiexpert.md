create procedure update_synapseiexpert(IN achievementcategoryid bigint, IN userids bigint[])
    language plpgsql
as
$$
BEGIN
    CALL update_subject_expert('Synapse I', achievementCategoryId, userIds);
END;
$$;

alter procedure update_synapseiexpert(bigint, bigint[]) owner to learner;
